# Implementation Plan: F07 — JSON Contracts and Schemas (`schemas/`)

## 1. Dependencies on Other Features

- **Depends on:** None (Foundation feature — implemented first so all agents and evals validate against these schemas).

## 2. Files to Create or Change

- `schemas/project.schema.json`
- `schemas/generation.schema.json`
- `schemas/review.schema.json`
- `schemas/failure-analysis.schema.json`
- `schemas/failure-repair.schema.json`
- `schemas/requirements.schema.json`

## 3. Ordered Tasks

### Task F07-T1: Create `project.schema.json`, `generation.schema.json`, and `review.schema.json`

- **Description:** Author JSON Schema Draft 2020-12 files for `project.json` (F01), `generation.json` / `generation.r<N>.json` (F02, F03), and `review.json` (F04) with `"additionalProperties": false` and required envelope fields (`schemaVersion`, `agent`, `createdAt`, `runId`).
- **Acceptance Criteria:**
  - [ ] Valid JSON Schema Draft 2020-12 syntax.
  - [ ] Includes the requirement traceability fields (`requirementsFile`, `scenarioId`, `unmappedScenarios`, `requirementCompliance`) from F07 Section 4.
  - [ ] Validates all complete JSON examples in `docs/features/F01-codebase-analyzer.md`, `F02-generator-generate.md`, `F03-generator-fix.md`, and `F04-reviewer.md`.
- **Files Touched:** `schemas/project.schema.json`, `schemas/generation.schema.json`, `schemas/review.schema.json`

### Task F07-T2: Create `failure-analysis.schema.json`, `failure-repair.schema.json`, and `requirements.schema.json`

- **Description:** Author JSON Schema Draft 2020-12 files for `failure-analysis.json` (F05), `failure-repair.json` (F06), and `requirements.json` (F13).
- **Acceptance Criteria:**
  - [ ] Validates all JSON examples in `docs/features/F05-failure-analyzer.md`, `F06-generator-repair-failures.md`, and `F13-requirement-analyzer.md`.
- **Files Touched:** `schemas/failure-analysis.schema.json`, `schemas/failure-repair.schema.json`, `schemas/requirements.schema.json`

## 4. How to Verify Manually

1. Validate all 6 schema files and the example payloads in `docs/features/F01..F06` and `F13` using `npx ajv-cli` or the `evals/` schema self-test.
