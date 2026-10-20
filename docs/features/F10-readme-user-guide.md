# F10: README and User Guide (`README.md`)

## 1. Purpose

Provide clear, concise documentation in `README.md` at the repository root so developers can understand the architecture, install the agents into their own repository for any of the four supported runtimes (Claude Code, OpenAI Codex CLI, Gemini CLI, GitHub Copilot), run both workflows (test generation/review and failure analysis/repair), and customize `.unit-test-agent/project.json`.

---

## 2. Invocation and Arguments

N/A (Documentation artifact at `README.md`).

---

## 3. Inputs and Outputs

### Inputs

- [docs/runtime-compatibility.md](../runtime-compatibility.md)
- [docs/decisions/ADR-001-packaging.md](../decisions/ADR-001-packaging.md)
- Feature specifications `F01`–`F09` and `F11`

### Output

- `README.md` at the repository root.

### Required Sections in `README.md`

1. **Overview & Key Principles:**
   - Pure Markdown agents, zero helper scripts in user repos, language-agnostic (JS/TS, Python, Java/Kotlin, C#/.NET, Go, Rust), strict guardrails (never touches production source files, never weakens assertions to hide bugs).
2. **Supported Runtimes:**
   - Quick comparison table of Claude Code, OpenAI Codex CLI, Gemini CLI, and GitHub Copilot (VS Code Agent Mode).
3. **Installation (Copy-Based, No Install Script):**
   - Exact `cp -r` commands for copying `agents/` and `schemas/` into `.unit-test-agent/` and copying the chosen runtime wrapper from `runtimes/<runtime>/`.
   - Recommendation on what to commit vs. ignore in `.gitignore` (commit `.unit-test-agent/project.json`, `.unit-test-agent/agents/`, `.unit-test-agent/schemas/`; optionally ignore `.unit-test-agent/runs/`).
4. **Workflow 1: Analyzing, Generating, Reviewing & Fixing Unit Tests:**
   - Step 1: `/codebase-analyzer` → `.unit-test-agent/project.json`
   - Step 2: `/unit-test-generator <targets>` → `.unit-test-agent/runs/<run-id>/generation.json`
   - Step 3: `/unit-test-reviewer <run-id>` → `.unit-test-agent/runs/<run-id>/review.json`
   - Step 4: `/unit-test-generator --fix <run-id>` → `.unit-test-agent/runs/<run-id>/generation.r1.json`
5. **Workflow 2: Regression Failure Analysis & Automated Test Repair:**
   - Step 1: `/unit-test-failure-analyzer [targets | --log <file>]` → `.unit-test-agent/runs/<run-id>/failure-analysis.json`
   - Step 2: `/unit-test-generator --repair-failures <run-id>` → `.unit-test-agent/runs/<run-id>/failure-repair.json`
6. **Customizing `project.json`:**
   - How to answer `missing` questions, customize `policy.coverageThresholds` (including 100% coverage) and `maxRepairAttempts`, and how user edits are preserved across re-runs.
7. **Handling `suspectedDefects`:**
   - How the agents report production bugs without corrupting test assertions.
8. **Contributor Guide & Running Evals (`evals/`):**
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
- [ ] Documents all 4 agents (`codebase-analyzer`, `unit-test-generator`, `unit-test-reviewer`, `unit-test-failure-analyzer`) and all generator modes (`generate`, `--fix`, `--repair-failures`).
- [ ] Explains `project.json` customization, `missing` resolution, and `suspectedDefects` handling.
- [ ] Explains how to run the fixture evaluations in `evals/`.
