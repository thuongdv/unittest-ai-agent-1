# Project Progress Tracker (`PROGRESS.md`)

> **Session Bootstrap Rule:** Any AI agent starting or resuming work in this repository MUST read this file first, check for `blocked` items or open questions, and pick up the next `todo` task in the **Master Task Table** below (ordered by feature dependencies).

---

## 1. Current Phase & Gate Status

- **Current Phase:** Step 3 Ready — **Specs & Plans Approved (amended 2026-11-05 with `F13` requirement analysis and `F14` run report); Ready to start Task 1 (`F07-T1`) in next session**
- **Completed Phases:**
  - [x] Step 0: Define scope including `unit-test-failure-analyzer` (F05) and `unit-test-generator --repair-failures` (F06)
  - [x] Step 1: Research 4 runtimes ([`docs/runtime-compatibility.md`](docs/runtime-compatibility.md)) and record packaging decision ([`docs/decisions/ADR-001-packaging.md`](docs/decisions/ADR-001-packaging.md))
  - [x] Step 2: Author feature specs (`F01`–`F11`), implementation plans (`F01`–`F11`), `PROGRESS.md`, `CLAUDE.md`, and `AGENTS.md` (Approved by user)
  - [x] Step 2 amendments: `F12` orchestrator (DEC-004), `F13` requirement analyzer (DEC-005), `F14` run reporter (DEC-006)

---

## 2. Master Task Table (Ordered by Implementation Dependency)

Implementation order builds the foundation contracts (`F07`) and test fixtures (`F09`) first, then the automated output verifier (`F11`), followed by the canonical agents (`F01`, `F13`, `F02`–`F06`, `F14`), the orchestrator (`F12`), multi-runtime wrappers (`F08`), and user guide (`F10`).

| Order | Feature     | Task ID            | Task Description                                                                          | Status | Spec & Plan Links                                                                       | Notes                                 |
| :---: | :---------- | :----------------- | :---------------------------------------------------------------------------------------- | :----: | :-------------------------------------------------------------------------------------- | :------------------------------------ |
|   1   | **F07**     | `F07-T1`           | Create `project.schema.json`, `generation.schema.json`, and `review.schema.json` (incl. requirement traceability fields) | `todo` | [Spec](docs/features/F07-json-contracts.md) · [Plan](docs/plans/F07-plan.md)            | Foundation schemas                    |
|   2   | **F07**     | `F07-T2`           | Create `failure-analysis.schema.json`, `failure-repair.schema.json`, and `requirements.schema.json` | `todo` | [Spec](docs/features/F07-json-contracts.md) · [Plan](docs/plans/F07-plan.md)            | Failure & requirement schemas         |
|   3   | **F09**     | `F09-T1`           | Create `examples/ts-vitest/` fixture project (deliberate bug, seeded failing tests, `discount.ts` + `requirements/SHOP-142.md`) | `todo` | [Spec](docs/features/F09-example-fixtures.md) · [Plan](docs/plans/F09-plan.md)          | TypeScript + Vitest fixture           |
|   4   | **F09**     | `F09-T2`           | Create `examples/py-pytest/` fixture project (incl. `restock.py` + `requirements/INV-77.md`) & `docs/manual-test-procedure.md` | `todo` | [Spec](docs/features/F09-example-fixtures.md) · [Plan](docs/plans/F09-plan.md)          | Python + pytest fixture               |
|   5   | **F11**     | `F11-T1`           | Scaffold `evals/` package, schema validator, loader & checks `CHK-01`–`CHK-05`            | `todo` | [Spec](docs/features/F11-automated-output-checks.md) · [Plan](docs/plans/F11-plan.md)   | Built right after F07 & F09           |
|   6   | **F11**     | `F11-T2`           | Implement checks `CHK-06`–`CHK-16`, pass-rate reporter & `evals` self-tests               | `todo` | [Spec](docs/features/F11-automated-output-checks.md) · [Plan](docs/plans/F11-plan.md)   | Verifies all 16 invariants            |
|   7   | **F01**     | `F01-T1`           | Write canonical `agents/codebase-analyzer.md`                                             | `todo` | [Spec](docs/features/F01-codebase-analyzer.md) · [Plan](docs/plans/F01-plan.md)         | 6-ecosystem tables + baseline         |
|   8   | **F13**     | `F13-T1`           | Build read-only signature extractors `tools/extract-signatures.mjs` & `tools/extract_signatures.py` + `CHK-16` tests | `todo` | [Spec](docs/features/F13-requirement-analyzer.md) · [Plan](docs/plans/F13-plan.md)      | Only scripts installed (DEC-007)      |
|   9   | **F13**     | `F13-T2`           | Write canonical `agents/unit-test-requirement-analyzer.md` (Jira MCP / `--req-file`, signatures only, no bodies) | `todo` | [Spec](docs/features/F13-requirement-analyzer.md) · [Plan](docs/plans/F13-plan.md)      | Isolated subagent; Jira read-only     |
|  10   | **F02**     | `F02-T1`           | Write canonical `agents/unit-test-generator.md` (Generate mode, `--requirements` scenario mapping, preconditions) | `todo` | [Spec](docs/features/F02-generator-generate.md) · [Plan](docs/plans/F02-plan.md)        | Includes `suspectedDefects` rules     |
|  11   | **F03**     | `F03-T1`           | Add Fix mode (`--fix <run>`) to `agents/unit-test-generator.md`                           | `todo` | [Spec](docs/features/F03-generator-fix.md) · [Plan](docs/plans/F03-plan.md)             | Writes `generation.r<N>.json`         |
|  12   | **F04**     | `F04-T1`           | Write canonical `agents/unit-test-reviewer.md` (9-point rubric, requirement compliance, mutation, 3x rerun) | `todo` | [Spec](docs/features/F04-reviewer.md) · [Plan](docs/plans/F04-plan.md)                  | Fresh-context review                  |
|  13   | **F05**     | `F05-T1`           | Write canonical `agents/unit-test-failure-analyzer.md` (direct run & `--log`)             | `todo` | [Spec](docs/features/F05-failure-analyzer.md) · [Plan](docs/plans/F05-plan.md)          | Root-cause classification             |
|  14   | **F06**     | `F06-T1`           | Add Repair-Failures mode (`--repair-failures <run>`) to `agents/unit-test-generator.md`   | `todo` | [Spec](docs/features/F06-generator-repair-failures.md) · [Plan](docs/plans/F06-plan.md) | Writes `failure-repair.json`          |
|  15   | **F14**     | `F14-T1`           | Write canonical `agents/unit-test-reporter.md` (`report.md` for developers & QA)          | `todo` | [Spec](docs/features/F14-reporter.md) · [Plan](docs/plans/F14-plan.md)                  | Renders only; recommendation table    |
|  16   | **F12**     | `F12-T1`           | Write canonical `agents/unit-test-orchestrator.md` (requirements → generate/review/fix loop → report & `--failures`) | `todo` | [Spec](docs/features/F12-orchestrator.md) · [Plan](docs/plans/F12-plan.md)              | Sequences only; no analyzer           |
|  17   | **F08**     | `F08-T1`           | Create thin wrappers (all agents + orchestrator) & isolated subagents for Claude Code and Codex CLI | `todo` | [Spec](docs/features/F08-multi-runtime-packaging.md) · [Plan](docs/plans/F08-plan.md)   | `runtimes/claude`, `runtimes/codex`; verify Jira MCP |
|  18   | **F08**     | `F08-T2`           | Create thin wrappers (all agents + orchestrator) & isolated subagents for Gemini CLI and GitHub Copilot | `todo` | [Spec](docs/features/F08-multi-runtime-packaging.md) · [Plan](docs/plans/F08-plan.md)   | `runtimes/gemini`, `runtimes/copilot`; verify Jira MCP |
|  19   | **F01–F06, F12–F14** | `F01-T2`..`F06-T2`, `F12-T2`, `F13-T3`, `F14-T2` | Run end-to-end verification of all 6 agents + orchestrator against fixtures & record `evals/` results | `todo` | [Manual Procedure](docs/features/F09-example-fixtures.md)                               | Updates Section 3 matrix below        |
|  20   | **F10**     | `F10-T1`           | Write comprehensive `README.md` user & contributor guide (incl. Jira MCP setup, reading `report.md`) | `todo` | [Spec](docs/features/F10-readme-user-guide.md) · [Plan](docs/plans/F10-plan.md)         | Final documentation pass              |

---

## 3. Per-Runtime Verification Matrix

Tracks end-to-end verification status (`untested` | `passing (pass_rate)` | `failing`) across fixtures (`ts-vitest`, `py-pytest`).

| Agent / Mode                                                             | Claude Code | OpenAI Codex CLI | Gemini CLI | GitHub Copilot (VS Code) |
| :----------------------------------------------------------------------- | :---------: | :--------------: | :--------: | :----------------------: |
| `codebase-analyzer` (`F01`)                                              | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Generate mode (`F02`)                            | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Generate mode with `--requirements` (`F02`)     | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Fix mode `--fix` (`F03`)                         | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-reviewer` (`F04`)                                             | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-failure-analyzer` (`F05`)                                     | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-generator` — Repair-Failures mode `--repair-failures` (`F06`) | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-requirement-analyzer` — `--jira` / `--req-file` (`F13`)      | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-reporter` (`F14`)                                             | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-orchestrator` — Generate & review workflow (`F12`)            | `untested`  |    `untested`    | `untested` |        `untested`        |
| `unit-test-orchestrator` — `--failures` workflow (`F12`)                 | `untested`  |    `untested`    | `untested` |        `untested`        |

---

## 4. Decisions Log

| Date       | ID          | Decision Summary                                                                                                                                                                                                   | Reference                                                              |
| :--------- | :---------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| 2026-10-20 | **DEC-001** | Added `unit-test-failure-analyzer` (`F05`) as a 4th standalone agent supporting both direct test execution (`[targets]`) and log ingestion (`--log <file>`).                                                       | [`F05`](docs/features/F05-failure-analyzer.md)                         |
| 2026-10-20 | **DEC-002** | Added `--repair-failures <run>` (`F06`) as a 3rd mode on `unit-test-generator` that modifies only test files, fixes stale/broken tests, and skips + logs `source_regression_bug` items in `suspectedDefects`.      | [`F06`](docs/features/F06-generator-repair-failures.md)                |
| 2026-10-20 | **ADR-001** | Adopted Canonical `agents/` + Thin Per-Runtime Wrappers (`runtimes/{claude,codex,gemini,copilot}`). Target repos store canonical files in `.unit-test-agent/{agents,schemas}` and copy their runtime's dot-folder. | [`ADR-001`](docs/decisions/ADR-001-packaging.md)                       |
| 2026-10-20 | **DEC-003** | Removed legacy root `unittest-ai.example.json` and initial bootstrap `docs/prompt.md` after decomposing all specs, guardrails, and plans into `docs/features/`, `docs/plans/`, and `AGENTS.md`.                    | [`F07`](docs/features/F07-json-contracts.md), [`AGENTS.md`](AGENTS.md) |
| 2026-10-22 | **DEC-004** | Replaced the "no orchestrator" constraint with `unit-test-orchestrator` (`F12`): one command per workflow, review/fix loop capped by `--max-review-rounds` (default 3), stops and points to `/codebase-analyzer` when `project.json` is missing. `codebase-analyzer` is one-time setup, never part of a workflow. Orchestrator writes no files and holds no agent rules. | [`F12`](docs/features/F12-orchestrator.md), [`AGENTS.md`](AGENTS.md) |
| 2026-11-05 | **DEC-005** | Added requirement analysis (`F13`, `unit-test-requirement-analyzer`) to address the test oracle problem: requirements come from Jira via MCP (`--jira`, primary, for service demos) or `--req-file` (offline/fixtures); they are optional, and without them the implementation is the requirement and reports say so. The analyzer runs in an isolated subagent, never reads `sourceRoots`/`testRoots` (per the reviewed article's Layer 3), and never writes to Jira. Generator maps scenarios to tests via `--requirements <run>`; reviewer classifies each requirement `pass`/`bug`/`test_issue`/`unclear`/`not_covered`; `bug` does not block test approval. | [`F13`](docs/features/F13-requirement-analyzer.md), [`F02`](docs/features/F02-generator-generate.md), [`F04`](docs/features/F04-reviewer.md) |
| 2026-11-05 | **DEC-006** | Added `unit-test-reporter` (`F14`): writes `runs/<run>/report.md` for developers, QA, and BA/PO (recommendation, scorecard, defects, traceability matrix, open questions). Render-only; runs as the last step of both orchestrator workflows so the orchestrator still writes no files. | [`F14`](docs/features/F14-reporter.md), [`F12`](docs/features/F12-orchestrator.md) |
| 2026-11-05 | **DEC-007** | Requirement analyzer may see public signatures (names, parameter and return types) but never bodies, default values, comments, or constant values; supersedes the code-blind part of DEC-005. Signatures come only from read-only extractors in `tools/` (JS/TS via the repo's `typescript`, Python via standard `ast`), the single exception to the no-scripts rule; other ecosystems use strict mode. `CHK-16` tests the extractors for leaks. | [`F13`](docs/features/F13-requirement-analyzer.md), [`AGENTS.md`](AGENTS.md), [`ADR-001`](docs/decisions/ADR-001-packaging.md) |

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
  - DEC-004: Added `unit-test-orchestrator` (`F12` spec + plan, task `F12-T1`); made `codebase-analyzer` one-time setup outside workflows; updated `AGENTS.md`, `CLAUDE.md`, ADR-001 (amendment), `F08`–`F11` specs/plans (orchestrator wrappers, fixture Step E, README sections, `CHK-12`), and the overview page.
- **2026-11-05:**
  - DEC-005: Added `F13` requirement analyzer (spec + plan); `F02` `--requirements` scenario mapping (`scenarioId`, `unmappedScenarios`); `F03` fixes for requirement findings; `F04` 9-point rubric and `requirementCompliance`; `F07` `requirements.schema.json` and traceability fields; `F09` requirement-violation fixtures (`discount.ts` + `SHOP-142.md`, `restock.py` + `INV-77.md`); `F11` `CHK-13`/`CHK-14`.
  - DEC-006: Added `F14` run reporter (spec + plan) and `CHK-15`; `F12` runs the requirement analyzer first and the reporter last.
  - Updated `F08` (wrappers + isolated requirement-analyzer subagent, Jira MCP access), `F10` (Jira setup, reading `report.md`), `AGENTS.md`, `CLAUDE.md`, ADR-001 (amendment), and the overview page.
  - DEC-007: Requirement analyzer sees signatures via read-only extractors (`tools/`); `F13` rewritten (targets input, `codeContract`, `call`/`expectedValue`, strict-mode fallback; tasks T1 extractors, T2 agent, T3 verify); `AGENTS.md` no-scripts exception; ADR-001 amendment; `F02`, `F07`, `F08`, `F09`, `F10`, `F11` (`CHK-16`), `F12`, and the overview page updated.
