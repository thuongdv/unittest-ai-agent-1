# Implementation Plan: F13 — Requirement Analyzer (`unit-test-requirement-analyzer`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/requirements.schema.json`)](../features/F07-json-contracts.md)
  - [F01: Codebase Analyzer (`project.json`)](../features/F01-codebase-analyzer.md)
  - [F09: Example Fixtures (`requirements/*.md`)](../features/F09-example-fixtures.md)

## 2. Files to Create or Change

- `agents/unit-test-requirement-analyzer.md` — Canonical Markdown agent instructions.
- Runtime wrappers and the isolated subagent are created in [F08](F08-plan.md).

## 3. Ordered Tasks

### Task F13-T1: Draft `agents/unit-test-requirement-analyzer.md`

- **Description:** Write the canonical instructions: argument routing (`--jira` vs `--req-file`), preconditions (including Jira MCP availability), the isolation & read-only rules table, allowed Jira fields, scenario design table (`happy_path`, `boundary`, `edge`, `error`), `UNC` rules, and `requirements.json` output rules.
- **Acceptance Criteria:**
  - [ ] Forbids reading `sourceRoots` and `testRoots` and forbids every Jira write operation.
  - [ ] Requires a verbatim `sourceQuote` per requirement and a `snapshot` per source.
  - [ ] Requires `UNC` items instead of guessed expected results.
  - [ ] Output matches `schemas/requirements.schema.json`.
- **Files Touched:** `agents/unit-test-requirement-analyzer.md`

### Task F13-T2: Verify on Fixtures

- **Description:** Run `/unit-test-requirement-analyzer --req-file requirements/SHOP-142.md` on `examples/ts-vitest` and `--req-file requirements/INV-77.md` on `examples/py-pytest`. Run `--jira SHOP-142` once against a demo Jira project.
- **Acceptance Criteria:**
  - [ ] `requirements.json` validates and has boundary scenarios at each stated threshold.
  - [ ] The fixture's planted ambiguity produces an `UNC` item.
  - [ ] The Jira issue is unchanged after the run (no new comments, edits, or transitions).
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. In a temp copy of `examples/ts-vitest` with `project.json`, run `/unit-test-requirement-analyzer --req-file requirements/SHOP-142.md`.
2. Confirm `runs/<run-id>/requirements.json` lists the $100 threshold with scenarios at $99.99 and $100, and an `UNC` item for the rounding question.
3. Inspect the runtime transcript and confirm no file under `src/` or `tests/` was opened.

## 5. Risks to Verify During F08

- **Jira MCP setup differs per runtime** (Claude Code `.mcp.json`, Codex `config.toml`, Gemini `settings.json`, VS Code `.vscode/mcp.json`). Verify each runtime can reach the Atlassian remote MCP server and document the exact setup in the README ([F10](F10-plan.md)).
- **Tool allow-lists:** Where a runtime lets a subagent restrict tools (Claude Code `tools:`, Copilot `tools:`), grant only Jira read tools plus file read/write for `.unit-test-agent/`.
