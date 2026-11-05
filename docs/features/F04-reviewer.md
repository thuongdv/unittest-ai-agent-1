# F04: Unit Test Reviewer (`unit-test-reviewer`)

## 1. Purpose

Independently audit a test generation run (`<run-id>`) in a **fresh, isolated context** (via subagent delegation where supported). Evaluate every generated test case against a 9-point quality rubric, check each requirement against its tests and the code (when the run has `requirements.json`), run target-scoped mutation testing (when configured), re-run the new tests 3 times to detect flakiness, and write `.unit-test-agent/runs/<run-id>/review.json`.

The reviewer is the only agent that sees the requirement, the tests, and the code together, so it decides whether a mismatch is a code bug, a test issue, or an unclear requirement.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation         | Headless / CLI Invocation                                  | Fresh-Context Mechanism                                                                     |
| :----------------- | :----------------------------- | :--------------------------------------------------------- | :------------------------------------------------------------------------------------------ |
| **Claude Code**    | `/unit-test-reviewer <run-id>` | `claude -p "/unit-test-reviewer 20261020-091500-pricing"`  | Slash command delegates to `.claude/agents/unit-test-reviewer.md` subagent                  |
| **Codex CLI**      | `$unit-test-reviewer <run-id>` | `codex exec "$unit-test-reviewer 20261020-091500-pricing"` | Spawns `.codex/agents/unit-test-reviewer.md` subagent or runs in fresh `codex exec` context |
| **Gemini CLI**     | `/unit-test-reviewer <run-id>` | `gemini -p "/unit-test-reviewer 20261020-091500-pricing"`  | Delegates to `@unit-test-reviewer` (`.gemini/agents/unit-test-reviewer.md`)                 |
| **GitHub Copilot** | `/unit-test-reviewer <run-id>` | N/A (VS Code Chat Agent mode)                              | Invokes `agent/runSubagent` targeting `.github/agents/unit-test-reviewer.agent.md`          |

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has supported `schemaVersion`, and has no `blocking: true` items in `missing`.
2. `.unit-test-agent/runs/<run-id>/generation.json` (or latest `generation.r<N>.json`) exists and has supported `schemaVersion`.
3. The generated test files and target source files exist on disk.
4. If the generation file's `requirementsFile` is not `null`, `runs/<run-id>/requirements.json` exists with a supported `schemaVersion`.

### Output

- **Path:** `.unit-test-agent/runs/<run-id>/review.json`
- **Schema:** `.unit-test-agent/schemas/review.schema.json` (defined in [F07](F07-json-contracts.md))

### Complete JSON Example (`.unit-test-agent/runs/20261020-091500-pricing/review.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "unit-test-reviewer",
  "runId": "20261020-091500-pricing",
  "reviewedGenerationFile": "generation.json",
  "createdAt": "2026-10-20T09:22:00Z",
  "verdict": "changes_requested",
  "summary": "Found 1 weak assertion in TC-001 and 1 surviving boundary mutant at src/pricing.ts:28. Flakiness check passed (3/3 runs).",
  "rubricFindings": [
    {
      "findingId": "REV-001",
      "caseId": "TC-001",
      "scenarioId": null,
      "testFile": "tests/pricing.test.ts",
      "rubricCategory": "weak_or_tautological_assertion",
      "severity": "major",
      "issue": "TC-001 asserts expect(result).toBeDefined() instead of verifying the computed total amount.",
      "suggestion": "Assert the exact numeric result: expect(result).toBeCloseTo(108.5, 2)."
    }
  ],
  "requirementCompliance": {
    "status": "not_assessed",
    "requirementsFile": null,
    "results": []
  },
  "mutationTesting": {
    "status": "executed",
    "tool": "stryker",
    "commandExecuted": "npx stryker run --mutate src/pricing.ts",
    "mutationScore": 85.7,
    "killed": 12,
    "survived": 2,
    "timeout": 0,
    "survivingMutants": [
      {
        "file": "src/pricing.ts",
        "line": 28,
        "mutator": "ConditionalExpression",
        "original": "quantity >= 100",
        "replacement": "quantity > 100",
        "suggestion": "Add a boundary test case with quantity === 100 to verify the bulk discount tier activates at 100."
      }
    ]
  },
  "flakinessCheck": {
    "reruns": 3,
    "commandExecuted": "npx vitest run tests/pricing.test.ts",
    "runsPassed": 3,
    "flakyTests": []
  }
}
```

### Requirement-Compliance Excerpt (`runs/20261105-101500-shop-142/review.json`)

```json
{
  "requirementCompliance": {
    "status": "assessed",
    "requirementsFile": "requirements.json",
    "results": [
      {
        "requirementId": "REQ-001",
        "classification": "bug",
        "scenarioIds": ["SCN-001", "SCN-002", "SCN-003", "SCN-004"],
        "testCaseIds": ["TC-001", "TC-002", "TC-003", "TC-004"],
        "defectIds": ["DEF-001", "DEF-002"],
        "unclearIds": ["UNC-001"],
        "note": "calculateDiscount applies the discount only above $200; SCN-002 and SCN-003 fail against the code and are skipped as defects."
      },
      {
        "requirementId": "REQ-002",
        "classification": "pass",
        "scenarioIds": ["SCN-005"],
        "testCaseIds": ["TC-005"],
        "defectIds": [],
        "unclearIds": [],
        "note": "Negative amounts throw RangeError as required."
      }
    ]
  }
}
```

---

## 4. Step-by-Step Behavior

1. **Check Preconditions:**
   - Confirm `project.json` and `.unit-test-agent/runs/<run-id>/generation.json` (or latest `generation.r<N>.json`) exist and have supported `schemaVersion` and zero `blocking: true` items in `missing`.
2. **Read Context in Fresh Session:**
   - Load `project.json`, the latest generation JSON for `<run-id>`, `requirements.json` (if referenced), the `exemplarTestFiles`, the target source files, and the generated test files.
   - Verify that every non-null `caseId` referenced in the review exists in the generation JSON's `testCases` array, and every non-null `scenarioId` exists in `requirements.json`.
3. **Evaluate Each Test Case Against the 9-Point Rubric:**
   - `missing_branch_or_edge_case`: Unchecked conditional branches, error paths, or boundary values in the target symbol.
   - `duplicate_case`: Two or more tests exercising the exact same code path and equivalence partition.
   - `implementation_detail_coupling`: Asserting private state or internal call counts when observable return values/state transitions suffice.
   - `over_mocking`: Mocking pure domain helpers or value objects unnecessarily.
   - `non_determinism`: Unseeded random generators, wall-clock `Date.now()`/`time.time()` without fake timers, real network/filesystem I/O, or shared mutable state across tests.
   - `weak_or_tautological_assertion`: `toBeDefined()`, `toBeTruthy()`, `assert True`, or asserting a mock returns what it was just configured to return.
   - `naming_and_style_mismatch`: Deviation from the conventions established in `exemplarTestFiles`.
   - `missing_requirement_scenario` (requirement runs only): A scenario has no linked test case and no `unmappedScenarios` entry, or its `unmappedScenarios` reason is wrong because a target symbol does implement it. Severity `major`.
   - `expected_value_mirrors_implementation`: An assertion's expected value matches what the code returns but contradicts its linked scenario, any other scenario, or the documented contract. Severity `critical`.
   - Each finding records `findingId`, `caseId` (an `id` in `generation.json`, or `null` when the finding is about a missing test), `scenarioId` (or `null`), `testFile`, `rubricCategory`, `severity` (`critical` | `major` | `minor`), `issue`, and `suggestion`.
4. **Assess Requirement Compliance** (when `requirementsFile` is not `null`; otherwise write `status: "not_assessed"`, `requirementsFile: null`, `results: []`):
   - For each `REQ`, collect its scenarios, linked test cases, `suspectedDefects`, and `unclear` items, and assign one `classification` (first match wins):

     | Classification | When                                                                                                  | Who acts         |
     | :------------- | :---------------------------------------------------------------------------------------------------- | :--------------- |
     | `bug`          | A linked test case is `skipped_defect`: the test matches the requirement but the code does not.       | Developer        |
     | `test_issue`   | A scenario has a `missing_requirement_scenario` or `expected_value_mirrors_implementation` finding, or a linked test `failed`. | Generator (`--fix`) |
     | `unclear`      | The requirement has no scenarios; every part of it is an `unclear` item.                              | BA / PO          |
     | `not_covered`  | Every scenario of the requirement is in `unmappedScenarios`.                                          | QA               |
     | `pass`         | Every scenario has a passing test that asserts the scenario's expected value.                         | —                |

   - Record `scenarioIds`, `testCaseIds`, `defectIds`, `unclearIds`, and a one-sentence `note`.
5. **Run Scoped Mutation Testing:**
   - Check `project.json` for `mutationTool.value` and `commands.mutation.value`.
   - **If configured:** Execute the mutation tool scoped strictly to the `targets` from `generation.json` (e.g., Stryker `--mutate`, `mutmut --paths-to-mutate`, PIT `-DtargetClasses`, `cargo mutants -f`). Record `status: "executed"`, `mutationScore`, and `survivingMutants`.
   - **If not configured (`null`):** Record `status: "skipped_not_configured"`, `mutationScore: null`, `survivingMutants: []`, and a note in `summary`. **Do not fail the review when no mutation tool is configured.**
6. **Run Flakiness Reruns (3x):**
   - Execute `commands.test.value` scoped to `testFilesWritten` **3 separate times**.
   - If any test passes on some runs and fails on others (or fails on any of the 3 runs when it wasn't marked `skipped_defect`), record it in `flakyTests` with the failure message and run numbers.
7. **Determine Verdict & Write Output:**
   - Set `verdict` to `"changes_requested"` if:
     - Any rubric finding has `severity: "critical"` or `"major"`, OR
     - `flakyTests` is non-empty, OR
     - Preventable `survivingMutants` exist on reachable, non-defective lines.
   - Otherwise, set `verdict` to `"approved"`.
   - Requirement classifications `bug`, `unclear`, and `not_covered` never block approval: the tests are correct, and these results are reported to people by `unit-test-reporter` ([F14](F14-reporter.md)).
   - Write `.unit-test-agent/runs/<run-id>/review.json`. Never modify source files or test files during review.

---

## 5. Edge Cases

- **No mutation tool installed:** `mutationTesting.status` is `"skipped_not_configured"`, `mutationScore` is `null`, and the reviewer does not error out.
- **Skipped defect tests (`skipped_defect`):** Reviewer verifies that the skipped test still contains a meaningful assertion matching the `suspectedDefects` entry and does not flag it as a failing test.
- **Read-only enforcement:** The reviewer never edits test files or source files; it only writes `review.json` (and cleans up any temporary mutation reports outside `.unit-test-agent/`).

---

## 6. Acceptance Criteria Checklist

- [ ] Runs in a fresh context (via subagent where supported by the runtime).
- [ ] Every non-null `caseId` in `rubricFindings` corresponds to an existing `id` in `generation.json` (or `generation.r<N>.json`), and every non-null `scenarioId` exists in `requirements.json`.
- [ ] Checks all 9 rubric categories and assigns `severity` (`critical` | `major` | `minor`), `issue`, and `suggestion`.
- [ ] With `requirements.json`: classifies every `REQ` as `bug`, `test_issue`, `unclear`, `not_covered`, or `pass`; without it: writes `requirementCompliance.status: "not_assessed"`.
- [ ] Runs target-scoped mutation testing if configured in `project.json`, or cleanly skips and reports `skipped_not_configured` if not configured.
- [ ] Executes the generated tests 3 consecutive times and reports any intermittent failures in `flakyTests`.
- [ ] Writes a valid `.unit-test-agent/runs/<run-id>/review.json` conforming to `schemas/review.schema.json`.
- [ ] Modifies zero source or test files (`git status` outside `.unit-test-agent/` remains clean).
