# Project Progress Tracker (`PROGRESS.md`)

> **Session Bootstrap Rule:** Any AI agent starting or resuming work in this repository MUST read this file first, check for `blocked` items or open questions, and pick up the next `todo` task in the **Master Task Table** below (ordered by feature dependencies).

---

## 1. Current Phase & Gate Status

- **Current Phase:** Step 3 Ready — **Specs & Plans Approved; Ready to start Task 1 (`F07-T1`) in next session**
- **Completed Phases:**
  - [x] Step 0: Define scope including `unit-test-failure-analyzer` (F05) and `unit-test-generator --repair-failures` (F06)
  - [x] Step 1: Research 4 runtimes ([`docs/runtime-compatibility.md`](docs/runtime-compatibility.md)) and record packaging decision ([`docs/decisions/ADR-001-packaging.md`](docs/decisions/ADR-001-packaging.md))
  - [x] Step 2: Author feature specs (`F01`–`F11`), implementation plans (`F01`–`F11`), `PROGRESS.md`, `CLAUDE.md`, and `AGENTS.md` (Approved by user)

---

## 2. Master Task Table (Ordered by Implementation Dependency)

Implementation order builds the foundation contracts (`F07`) and test fixtures (`F09`) first, then the automated output verifier (`F11`), followed by the canonical agents (`F01`–`F06`), multi-runtime wrappers (`F08`), and user guide (`F10`).

| Order | Feature     | Task ID            | Task Description                                                                          | Status | Spec & Plan Links                                                                       | Notes                                 |
| :---: | :---------- | :----------------- | :---------------------------------------------------------------------------------------- | :----: | :-------------------------------------------------------------------------------------- | :------------------------------------ |
|   1   | **F07**     | `F07-T1`           | Create `project.schema.json`, `generation.schema.json`, and `review.schema.json`          | `todo` | [Spec](docs/features/F07-json-contracts.md) · [Plan](docs/plans/F07-plan.md)            | Foundation schemas                    |
|   2   | **F07**     | `F07-T2`           | Create `failure-analysis.schema.json` and `failure-repair.schema.json`                    | `todo` | [Spec](docs/features/F07-json-contracts.md) · [Plan](docs/plans/F07-plan.md)            | Failure workflow schemas              |
|   3   | **F09**     | `F09-T1`           | Create `examples/ts-vitest/` fixture project (with deliberate bug + seeded failing tests) | `todo` | [Spec](docs/features/F09-example-fixtures.md) · [Plan](docs/plans/F09-plan.md)          | TypeScript + Vitest fixture           |
|   4   | **F09**     | `F09-T2`           | Create `examples/py-pytest/` fixture project & `docs/manual-test-procedure.md`            | `todo` | [Spec](docs/features/F09-example-fixtures.md) · [Plan](docs/plans/F09-plan.md)          | Python + pytest fixture               |
|   5   | **F11**     | `F11-T1`           | Scaffold `evals/` package, schema validator, loader & checks `CHK-01`–`CHK-05`            | `todo` | [Spec](docs/features/F11-automated-output-checks.md) · [Plan](docs/plans/F11-plan.md)   | Built right after F07 & F09           |
|   6   | **F11**     | `F11-T2`           | Implement checks `CHK-06`–`CHK-11`, pass-rate reporter & `evals` self-tests               | `todo` | [Spec](docs/features/F11-automated-output-checks.md) · [Plan](docs/plans/F11-plan.md)   | Verifies all 11 invariants            |
|   7   | **F01**     | `F01-T1`           | Write canonical `agents/codebase-analyzer.md`                                             | `todo` | [Spec](docs/features/F01-codebase-analyzer.md) · [Plan](docs/plans/F01-plan.md)         | 6-ecosystem tables + baseline         |
|   8   | **F02**     | `F02-T1`           | Write canonical `agents/unit-test-generator.md` (Generate mode + preconditions)           | `todo` | [Spec](docs/features/F02-generator-generate.md) · [Plan](docs/plans/F02-plan.md)        | Includes `suspectedDefects` rules     |
|   9   | **F03**     | `F03-T1`           | Add Fix mode (`--fix <run>`) to `agents/unit-test-generator.md`                           | `todo` | [Spec](docs/features/F03-generator-fix.md) · [Plan](docs/plans/F03-plan.md)             | Writes `generation.r<N>.json`         |
|  10   | **F04**     | `F04-T1`           | Write canonical `agents/unit-test-reviewer.md` (7-point rubric, mutation, 3x rerun)       | `todo` | [Spec](docs/features/F04-reviewer.md) · [Plan](docs/plans/F04-plan.md)                  | Fresh-context review                  |
|  11   | **F05**     | `F05-T1`           | Write canonical `agents/unit-test-failure-analyzer.md` (direct run & `--log`)             | `todo` | [Spec](docs/features/F05-failure-analyzer.md) · [Plan](docs/plans/F05-plan.md)          | Root-cause classification             |
|  12   | **F06**     | `F06-T1`           | Add Repair-Failures mode (`--repair-failures <run>`) to `agents/unit-test-generator.md`   | `todo` | [Spec](docs/features/F06-generator-repair-failures.md) · [Plan](docs/plans/F06-plan.md) | Writes `failure-repair.json`          |
|  13   | **F08**     | `F08-T1`           | Create thin wrappers & reviewer subagents for Claude Code and Codex CLI                   | `todo` | [Spec](docs/features/F08-multi-runtime-packaging.md) · [Plan](docs/plans/F08-plan.md)   | `runtimes/claude`, `runtimes/codex`   |
|  14   | **F08**     | `F08-T2`           | Create thin wrappers & reviewer subagents for Gemini CLI and GitHub Copilot               | `todo` | [Spec](docs/features/F08-multi-runtime-packaging.md) · [Plan](docs/plans/F08-plan.md)   | `runtimes/gemini`, `runtimes/copilot` |
|  15   | **F01–F06** | `F01-T2`..`F06-T2` | Run end-to-end verification of all 4 agents against fixtures & record `evals/` results    | `todo` | [Manual Procedure](docs/features/F09-example-fixtures.md)                               | Updates Section 3 matrix below        |
|  16   | **F10**     | `F10-T1`           | Write comprehensive `README.md` user & contributor guide                                  | `todo` | [Spec](docs/features/F10-readme-user-guide.md) · [Plan](docs/plans/F10-plan.md)         | Final documentation pass              |

---

## 3. Per-Runtime Verification Matrix

Tracks end-to-end verification status (`untested` | `passing (pass_rate)` | `failing`) across fixtures (`ts-vitest`, `py-pytest`).

| Agent / Mode                                                             | Claude Code | OpenAI Codex CLI | Gemini CLI | GitHub Copilot (VS Code) |
| :----------------------------------------------------------------------- | :---------: | :--------------: | :--------: | :----------------------: |
| `codebase-analyzer` (`F01`)                                              | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Generate mode (`F02`)                            | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Fix mode `--fix` (`F03`)                         | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-reviewer` (`F04`)                                             | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-failure-analyzer` (`F05`)                                     | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Repair-Failures mode `--repair-failures` (`F06`) | `untested`  |    `untested`    | `untested` |        `untested`        |

---

## 4. Decisions Log

| Date       | ID          | Decision Summary                                                                                                                                                                                                   | Reference                                                              |
| :--------- | :---------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| 2026-10-20 | **DEC-001** | Added `unit-test-failure-analyzer` (`F05`) as a 4th standalone agent supporting both direct test execution (`[targets]`) and log ingestion (`--log <file>`).                                                       | [`F05`](docs/features/F05-failure-analyzer.md)                         |
| 2026-10-20 | **DEC-002** | Added `--repair-failures <run>` (`F06`) as a 3rd mode on `unit-test-generator` that modifies only test files, fixes stale/broken tests, and skips + logs `source_regression_bug` items in `suspectedDefects`.      | [`F06`](docs/features/F06-generator-repair-failures.md)                |
| 2026-10-20 | **ADR-001** | Adopted Canonical `agents/` + Thin Per-Runtime Wrappers (`runtimes/{claude,codex,gemini,copilot}`). Target repos store canonical files in `.unit-test-agent/{agents,schemas}` and copy their runtime's dot-folder. | [`ADR-001`](docs/decisions/ADR-001-packaging.md)                       |
| 2026-10-20 | **DEC-003** | Removed legacy root `unittest-ai.example.json` and initial bootstrap `docs/prompt.md` after decomposing all specs, guardrails, and plans into `docs/features/`, `docs/plans/`, and `AGENTS.md`.                    | [`F07`](docs/features/F07-json-contracts.md), [`AGENTS.md`](AGENTS.md) |

---

## 5. Open Questions

_(None — all questions resolved.)_

---

## 6. Changelog

- **2026-10-20:**
  - Defined scope for all 4 agents including `unit-test-failure-analyzer` (`F05`) and `unit-test-generator --repair-failures` (`F06`).
  - Completed Step 1 runtime research in [`docs/runtime-compatibility.md`](docs/runtime-compatibility.md) and [`docs/decisions/ADR-001-packaging.md`](docs/decisions/ADR-001-packaging.md).
  - Completed Step 2 specifications (`docs/features/F01..F11`), implementation plans (`docs/plans/F01..F11`), `PROGRESS.md`, `CLAUDE.md`, and `AGENTS.md`.
  - Approved `ADR-001` packaging architecture, removed legacy `unittest-ai.example.json` and bootstrap `docs/prompt.md`, and marked project ready for Step 3 implementation.
- **2026-10-22:**
  - Added high-level user-facing feature overview [`docs/users/features-overview.html`](docs/users/features-overview.html) (workflows, agents, guardrails, failure triage, runtimes, install, QA, roadmap) with Mermaid diagrams.
