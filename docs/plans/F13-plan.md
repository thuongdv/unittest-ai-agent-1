# Implementation Plan: F13 — Requirement Analyzer (`unit-test-requirement-analyzer`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/requirements.schema.json`)](../features/F07-json-contracts.md)
  - [F01: Codebase Analyzer (`project.json`)](../features/F01-codebase-analyzer.md)
  - [F09: Example Fixtures (`requirements/*.md`, `discount.ts`, `restock.py`)](../features/F09-example-fixtures.md)
  - [F11: Automated Output Checks (`CHK-16` extractor tests)](../features/F11-automated-output-checks.md)

## 2. Files to Create or Change

- `tools/extract-signatures.mjs` — TS/JS signature extractor (uses the target repo's `typescript`).
- `tools/extract_signatures.py` — Python signature extractor (standard-library `ast`).
- `agents/unit-test-requirement-analyzer.md` — Canonical Markdown agent instructions.
- `evals/tests/extractors.test.ts` — Extractor unit tests (`CHK-16`), run against `examples/` source files.
- Runtime wrappers and the isolated subagent are created in [F08](F08-plan.md).

## 3. Ordered Tasks

### Task F13-T1: Build the Signature Extractors

- **Description:** Write `tools/extract-signatures.mjs` and `tools/extract_signatures.py` per F13 Section 4 (keep/drop rules, stdout JSON, exit code `2` when the parser is unavailable), plus `evals/tests/extractors.test.ts`.
- **Acceptance Criteria:**
  - [ ] On `examples/ts-vitest/src/discount.ts` and `examples/py-pytest/src/inventory/restock.py`, output contains the public signatures and none of: bodies, default values, comments/docstrings, constant or enum values (the thresholds `200` / `0.1` do not appear).
  - [ ] Exit code `2` with a one-line stderr message when `typescript` is not resolvable.
  - [ ] No dependencies beyond the target repo's runtime and installed packages; no file writes; no network.
- **Files Touched:** `tools/extract-signatures.mjs`, `tools/extract_signatures.py`, `evals/tests/extractors.test.ts`

### Task F13-T2: Draft `agents/unit-test-requirement-analyzer.md`

- **Description:** Write the canonical instructions: argument routing (`[targets]`, `--jira` vs `--req-file`), preconditions (including Jira MCP availability), the isolation & read-only rules table, the extractor table and strict-mode fallback, allowed Jira fields, scenario design table, `call`/`expectedValue` rules, `UNC` rules, and `requirements.json` output rules.
- **Acceptance Criteria:**
  - [ ] Forbids opening `sourceRoots` and `testRoots`, forbids values taken from names, and forbids every Jira write operation.
  - [ ] Requires a verbatim `sourceQuote` per requirement and a `snapshot` per source.
  - [ ] Requires `UNC` items instead of guessed expected results.
  - [ ] Output matches `schemas/requirements.schema.json`.
- **Files Touched:** `agents/unit-test-requirement-analyzer.md`

### Task F13-T3: Verify on Fixtures

- **Description:** Run `/unit-test-requirement-analyzer src/discount.ts --req-file requirements/SHOP-142.md` on `examples/ts-vitest` and `src/inventory/restock.py --req-file requirements/INV-77.md` on `examples/py-pytest`. Run `--jira SHOP-142` once against a demo Jira project.
- **Acceptance Criteria:**
  - [ ] `requirements.json` validates, has `codeContract.status: "extracted"`, and has boundary scenarios with `call`/`expectedValue` at each stated threshold.
  - [ ] Without targets, `codeContract.status` is `"not_requested"` and every `call` is `null`.
  - [ ] The fixture's planted ambiguity produces an `UNC` item.
  - [ ] The Jira issue is unchanged after the run (no new comments, edits, or transitions).
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. In a temp copy of `examples/ts-vitest` with `project.json`, run `node .unit-test-agent/tools/extract-signatures.mjs src/discount.ts` and confirm the output shows `calculateDiscount(amount: number): number` and no `200`.
2. Run `/unit-test-requirement-analyzer src/discount.ts --req-file requirements/SHOP-142.md`.
3. Confirm `runs/<run-id>/requirements.json` has `calculateDiscount(100)` → `10` and an `UNC` item for rounding.
4. Inspect the runtime transcript and confirm no file under `src/` or `tests/` was opened (only the extractor command ran).

## 5. Risks to Verify During F08

- **Jira MCP setup differs per runtime** (Claude Code `.mcp.json`, Codex `config.toml`, Gemini `settings.json`, VS Code `.vscode/mcp.json`). Verify each runtime can reach the Atlassian remote MCP server and document the exact setup in the README ([F10](F10-plan.md)).
- **Tool allow-lists:** Where a runtime lets a subagent restrict tools (Claude Code `tools:`, Copilot `tools:`), grant only Jira read tools, the shell (for the extractor command), and file read/write for `.unit-test-agent/`. A shell grant technically allows `cat src/...`; the canonical rule forbids it, and the transcript check in Section 4 verifies it.
