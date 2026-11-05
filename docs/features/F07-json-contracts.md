# F07: JSON Contracts and Schemas (`schemas/`)

## 1. Purpose

Define strict, human-reviewable JSON Schema (Draft 2020-12) files in `schemas/` for every JSON artifact written by the agents under `.unit-test-agent/`. These schemas serve both as the explicit contract included in agent instructions and as the validation rules executed by `evals/` ([F11](F11-automated-output-checks.md)).

---

## 2. Invocation and Arguments

Not invoked directly by users as a command. Schemas are:

- Read by agents from `.unit-test-agent/schemas/*.schema.json` to self-validate output structure.
- Validated automatically by `evals/` ([F11](F11-automated-output-checks.md)) using Ajv (JSON Schema Draft 2020-12).

---

## 3. Inputs and Outputs

### Schema Files Delivered in `schemas/`

1. `schemas/project.schema.json` — Validates `.unit-test-agent/project.json` ([F01](F01-codebase-analyzer.md))
2. `schemas/generation.schema.json` — Validates `.unit-test-agent/runs/<run-id>/generation.json` ([F02](F02-generator-generate.md)) and `generation.r<N>.json` ([F03](F03-generator-fix.md))
3. `schemas/review.schema.json` — Validates `.unit-test-agent/runs/<run-id>/review.json` ([F04](F04-reviewer.md))
4. `schemas/failure-analysis.schema.json` — Validates `.unit-test-agent/runs/<run-id>/failure-analysis.json` ([F05](F05-failure-analyzer.md))
5. `schemas/failure-repair.schema.json` — Validates `.unit-test-agent/runs/<run-id>/failure-repair.json` ([F06](F06-generator-repair-failures.md))
6. `schemas/requirements.schema.json` — Validates `.unit-test-agent/runs/<run-id>/requirements.json` ([F13](F13-requirement-analyzer.md))

`report.md` ([F14](F14-reporter.md)) is Markdown, not JSON, and has no schema; `evals/` checks it with `CHK-15`.

### Common Envelope Fields (Required Across All Outputs)

Every schema enforces:

- `schemaVersion`: `"1.0.0"` (semver pattern `^1\.[0-9]+\.[0-9]+$`)
- `agent`: `"codebase-analyzer"` | `"unit-test-generator"` | `"unit-test-reviewer"` | `"unit-test-failure-analyzer"` | `"unit-test-requirement-analyzer"`
- `createdAt`: ISO 8601 date-time string
- `runId`: Non-empty string (required on all run-scoped files: `generation`, `review`, `failure-analysis`, `failure-repair`, `requirements`)

### Complete JSON Schema Example (`schemas/failure-analysis.schema.json`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://unittest-ai-agent.local/schemas/failure-analysis.schema.json",
  "title": "UnitTestFailureAnalysis",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "schemaVersion",
    "agent",
    "runId",
    "createdAt",
    "inputSource",
    "summary",
    "failures"
  ],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "const": "1.0.0"
    },
    "agent": {
      "type": "string",
      "const": "unit-test-failure-analyzer"
    },
    "runId": {
      "type": "string",
      "minLength": 1
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    },
    "inputSource": {
      "type": "object",
      "additionalProperties": false,
      "required": ["mode", "commandExecuted", "logFile", "targets"],
      "properties": {
        "mode": { "type": "string", "enum": ["command", "logFile"] },
        "commandExecuted": { "type": ["string", "null"] },
        "logFile": { "type": ["string", "null"] },
        "targets": { "type": "array", "items": { "type": "string" } }
      }
    },
    "summary": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "totalTestsRun",
        "passed",
        "failed",
        "skipped",
        "analyzedFailures",
        "byCategory"
      ],
      "properties": {
        "totalTestsRun": { "type": "integer", "minimum": 0 },
        "passed": { "type": "integer", "minimum": 0 },
        "failed": { "type": "integer", "minimum": 0 },
        "skipped": { "type": "integer", "minimum": 0 },
        "analyzedFailures": { "type": "integer", "minimum": 0 },
        "byCategory": {
          "type": "object",
          "additionalProperties": false,
          "required": [
            "stale_test",
            "broken_mock_or_fixture",
            "source_regression_bug",
            "flaky_or_nondeterministic",
            "environment_or_dependency",
            "syntax_or_import_error"
          ],
          "properties": {
            "stale_test": { "type": "integer", "minimum": 0 },
            "broken_mock_or_fixture": { "type": "integer", "minimum": 0 },
            "source_regression_bug": { "type": "integer", "minimum": 0 },
            "flaky_or_nondeterministic": { "type": "integer", "minimum": 0 },
            "environment_or_dependency": { "type": "integer", "minimum": 0 },
            "syntax_or_import_error": { "type": "integer", "minimum": 0 }
          }
        }
      }
    },
    "failures": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "failureId",
          "testFile",
          "testName",
          "targetFile",
          "targetSymbol",
          "errorMessage",
          "stackTraceLocation",
          "category",
          "confidence",
          "rootCause",
          "evidence",
          "recommendedAction",
          "proposedChangeSummary"
        ],
        "properties": {
          "failureId": { "type": "string", "pattern": "^FAIL-[0-9]{3,}$" },
          "testFile": { "type": "string", "minLength": 1 },
          "testName": { "type": "string", "minLength": 1 },
          "targetFile": { "type": ["string", "null"] },
          "targetSymbol": { "type": ["string", "null"] },
          "errorMessage": { "type": "string" },
          "stackTraceLocation": { "type": ["string", "null"] },
          "category": {
            "type": "string",
            "enum": [
              "stale_test",
              "broken_mock_or_fixture",
              "source_regression_bug",
              "flaky_or_nondeterministic",
              "environment_or_dependency",
              "syntax_or_import_error"
            ]
          },
          "confidence": {
            "type": "string",
            "enum": ["high", "medium", "low"]
          },
          "rootCause": { "type": "string", "minLength": 1 },
          "evidence": {
            "type": "array",
            "minItems": 1,
            "items": { "type": "string" }
          },
          "recommendedAction": {
            "type": "string",
            "enum": [
              "update_test",
              "fix_mock_or_setup",
              "stabilize_test",
              "skip_and_report_defect",
              "fix_environment"
            ]
          },
          "proposedChangeSummary": { "type": "string", "minLength": 1 }
        }
      }
    }
  }
}
```

---

## 4. Step-by-Step Behavior

1. **Reusable `$defs` per Schema:**
   - In `project.schema.json`: Define `$defs/detectedString`, `$defs/detectedStringArray`, and `$defs/missingItem` so every detected field consistently enforces `{ "value": ..., "confidence": "high"|"medium"|"low", "evidence": "..." }`.
   - In `generation.schema.json` and `failure-repair.schema.json`: Share a consistent `$defs/suspectedDefect` structure (`defectId`, `targetFile`, `targetSymbol`, `line`, `scenarioId` (string or `null`), `testFile`, `description`, `expectedBehavior`, `actualBehavior`).
2. **Strict Validation (`additionalProperties: false`):**
   - Set `"additionalProperties": false` on all objects so typos in field names made by an agent are immediately caught by schema validation.
3. **Mode Discrimination in `generation.schema.json`:**
   - Support both `mode: "generate"` (`generation.json`) and `mode: "fix"` (`generation.r<N>.json`, which additionally requires `revision`, `parentGenerationFile`, `reviewFile`, and `addressedFindings`).
4. **Requirement Traceability Fields:**

   | Schema                       | Field                                    | Type                                                                                       |
   | :--------------------------- | :--------------------------------------- | :----------------------------------------------------------------------------------------- |
   | `requirements.schema.json`   | `sources[]`                              | `kind` (`jira` \| `file`), `ref`, `title`, `fieldsRead`, `fetchedAt`, `snapshot`           |
   | `requirements.schema.json`   | `requirements[]`                         | `id` (`^REQ-[0-9]{3,}$`), `sourceRef`, `sourceQuote`, `statement`, `scenarios[]`           |
   | `requirements.schema.json`   | `requirements[].scenarios[]`             | `id` (`^SCN-[0-9]{3,}$`), `category` (`happy_path` \| `edge` \| `boundary` \| `error`), `given`, `expected`, `call` and `expectedValue` (string or `null`) |
   | `requirements.schema.json`   | `codeContract`                           | `status` (`extracted` \| `unavailable` \| `not_requested`), `targets`, `commandExecuted`, `reason`, `symbols[]` (`file`, `name`, `kind`, `signature`) |
   | `requirements.schema.json`   | `unclear[]`                              | `id` (`^UNC-[0-9]{3,}$`), `sourceRef`, `requirementIds`, `question`, `reason`              |
   | `generation.schema.json`     | `requirementsFile`                       | `"requirements.json"` or `null` (required)                                                 |
   | `generation.schema.json`     | `testCases[].scenarioId`                 | `^SCN-[0-9]{3,}$` or `null` (required)                                                     |
   | `generation.schema.json`     | `unmappedScenarios[]`                    | `scenarioId`, `reason` (required; may be empty)                                            |
   | `review.schema.json`         | `rubricFindings[].caseId` / `.scenarioId` | string or `null` (both required)                                                          |
   | `review.schema.json`         | `rubricFindings[].rubricCategory`        | adds `missing_requirement_scenario`, `expected_value_mirrors_implementation`               |
   | `review.schema.json`         | `requirementCompliance`                  | `status` (`assessed` \| `not_assessed`), `requirementsFile`, `results[]` with `requirementId`, `classification` (`pass` \| `bug` \| `test_issue` \| `unclear` \| `not_covered`), `scenarioIds`, `testCaseIds`, `defectIds`, `unclearIds`, `note` |

---

## 5. Edge Cases

- **Nullable fields in `project.json`:** Undetected optional or missing fields must allow `"value": null` in `project.schema.json` while still requiring `confidence` and `evidence`.
- **Requirement compliance not assessed:** When `requirementCompliance.status` is `"not_assessed"`, `requirementsFile` must be `null` and `results` must be empty.
- **Skipped mutation testing in `review.json`:** When `mutationTesting.status` is `"skipped_not_configured"`, `mutationScore`, `killed`, `survived`, and `timeout` must allow `null`.

---

## 6. Acceptance Criteria Checklist

- [ ] All 6 JSON Schema files exist in `schemas/` (`project`, `generation`, `review`, `failure-analysis`, `failure-repair`, `requirements`).
- [ ] Every schema uses JSON Schema Draft 2020-12 and sets `"additionalProperties": false`.
- [ ] All complete JSON examples in `docs/features/F01` through `F06` and `F13` validate cleanly against their corresponding schema in `schemas/` (sections labelled "Excerpt" are partial and exempt).
- [ ] Every schema requires `schemaVersion`, `agent`, `createdAt`, and (for run files) `runId`.
