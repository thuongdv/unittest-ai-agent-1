# F10: README and User Guide (`README.md`)

## 1. Purpose

Provide clear, concise documentation in `README.md` at the repository root so developers can understand the architecture, install the agents into their own repository for any of the four supported runtimes (Claude Code, OpenAI Codex CLI, Gemini CLI, GitHub Copilot), run both workflows (test generation/review and failure analysis/repair), connect Jira requirements, read the run report, and customize `.unit-test-agent/project.json`.

---

## 2. Invocation and Arguments

N/A (Documentation artifact at `README.md`).

---

## 3. Inputs and Outputs

### Inputs

- [docs/runtime-compatibility.md](../runtime-compatibility.md)
- [docs/decisions/ADR-001-packaging.md](../decisions/ADR-001-packaging.md)
- Feature specifications `F01`–`F09` and `F11`–`F14`

### Output

- `README.md` at the repository root.

### Required Sections in `README.md`

1. **Overview & Key Principles:**
   - Markdown agents; the only scripts installed are the read-only signature extractors in `.unit-test-agent/tools/`, language-agnostic (JS/TS, Python, Java/Kotlin, C#/.NET, Go, Rust), strict guardrails (never touches production source files, never weakens assertions to hide bugs).
2. **Supported Runtimes:**
   - Quick comparison table of Claude Code, OpenAI Codex CLI, Gemini CLI, and GitHub Copilot (VS Code Agent Mode).
3. **Installation (Copy-Based, No Install Script):**
   - Exact `cp -r` commands for copying `agents/`, `schemas/`, and `tools/` into `.unit-test-agent/` and copying the chosen runtime wrapper from `runtimes/<runtime>/`.
   - Recommendation on what to commit vs. ignore in `.gitignore` (commit `.unit-test-agent/project.json`, `.unit-test-agent/agents/`, `.unit-test-agent/schemas/`; optionally ignore `.unit-test-agent/runs/`).
4. **One-Time Setup:**
   - `/codebase-analyzer` → `.unit-test-agent/project.json`. Re-run only when the repository's tooling changes.
5. **Workflow 1: Generating, Reviewing & Fixing Unit Tests:**
   - One command: `/unit-test-orchestrator <targets> [--jira <keys> | --req-file <files>] [--max-review-rounds N]` ([F12](F12-orchestrator.md)).
   - Equivalent manual steps: optional `/unit-test-requirement-analyzer --jira <keys>` → `requirements.json`; `/unit-test-generator <targets> [--requirements <run-id>]` → `generation.json`; `/unit-test-reviewer <run-id>` → `review.json`; `/unit-test-generator --fix <run-id>` → `generation.r1.json`; `/unit-test-reporter <run-id>` → `report.md`.
   - Why requirements matter: without them, tests are derived from the implementation and confirm what the code does, not what the business asked for.
6. **Workflow 2: Regression Failure Analysis & Automated Test Repair:**
   - One command: `/unit-test-orchestrator --failures [targets | --log <file>]`.
   - Equivalent manual steps: `/unit-test-failure-analyzer [targets | --log <file>]` → `failure-analysis.json`; `/unit-test-generator --repair-failures <run-id>` → `failure-repair.json`; `/unit-test-reporter <run-id>` → `report.md`.
7. **Requirements from Jira ([F13](F13-requirement-analyzer.md)):**
   - Per-runtime setup of a Jira MCP server (Claude Code, Codex CLI, Gemini CLI, VS Code Copilot), using the steps verified in [F08](F08-multi-runtime-packaging.md).
   - Which Jira fields are read, that the agent never writes to Jira, and that it sees code only as extracted signatures (no bodies); which ecosystems have an extractor and what strict mode means for the others.
   - `--req-file` for offline use, and how to read `UNC` questions.
8. **Reading the Run Report ([F14](F14-reporter.md)):**
   - Where `report.md` lives, the recommendation table, the developer and QA sections, and posting it to a pull request from CI.
9. **Customizing `project.json`:**
   - How to answer `missing` questions, customize `policy.coverageThresholds` (including 100% coverage) and `maxRepairAttempts`, and how user edits are preserved across re-runs.
10. **Handling `suspectedDefects`:**
   - How the agents report production bugs without corrupting test assertions.
11. **Contributor Guide & Running Evals (`evals/`):**
   - How contributors test changes against `examples/ts-vitest` and `examples/py-pytest` using the `evals/` Vitest suite.

---

## 4. Step-by-Step Behavior

1. Draft `README.md` with copy-pasteable commands for every runtime.
2. Verify all internal links to `docs/`, `schemas/`, `agents/`, `runtimes/`, and `examples/` resolve accurately.

---

## 5. Edge Cases

- **Users using multiple runtimes on the same team:** Document how a team can commit wrappers for multiple runtimes side-by-side (e.g., `.claude/`, `.gemini/`, `.github/`) all pointing to the same `.unit-test-agent/agents/` directory.

---

## 6. Acceptance Criteria Checklist

- [ ] `README.md` covers copy-based installation for all 4 runtimes.
- [ ] Documents all 6 agents (`codebase-analyzer`, `unit-test-generator`, `unit-test-reviewer`, `unit-test-failure-analyzer`, `unit-test-requirement-analyzer`, `unit-test-reporter`), all generator modes (`generate`, `--fix`, `--repair-failures`, with and without `--requirements`), and the `unit-test-orchestrator`.
- [ ] Documents Jira MCP setup for all 4 runtimes and how to read `report.md`.
- [ ] Presents `/codebase-analyzer` as one-time setup, not as a workflow step.
- [ ] Explains `project.json` customization, `missing` resolution, and `suspectedDefects` handling.
- [ ] Explains how to run the fixture evaluations in `evals/`.
