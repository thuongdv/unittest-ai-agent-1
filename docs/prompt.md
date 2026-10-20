# Project: Unit Test AI Agents (v1)

## Goal

Build a set of AI agents, written purely as Markdown instruction files, that analyze a codebase and then generate and review unit tests for it. They must work for any programming language and test framework. Users install the agents into their own repo and invoke them manually.

The agents must work in four runtimes:

- Claude Code
- OpenAI Codex CLI
- Gemini CLI
- GitHub Copilot (VS Code agent mode)

## Hard constraints

- **No helper code.** No JavaScript, TypeScript, Python or shell scripts for the agents themselves. Deliverables are:
  - `.md` instruction files
  - JSON examples and JSON Schema files
  - the thinnest wrapper/config file a runtime strictly requires (e.g. TOML, if Gemini CLI needs it)

  Exception: dev tooling that tests this repo (`evals/`, see F11) may be code. It is never installed into a user's repo and the agents never call it.

- **Agents use the target repo's tooling.** They do their work by running the target repo's own commands (install, test, coverage, mutation).
- **No orchestrator.** The user chains the agents by hand:
  - **Generation & review flow:** `/codebase-analyzer` → `/unit-test-generator <targets>` → `/unit-test-reviewer <run>` → `/unit-test-generator --fix <run>` (only if changes were requested).
  - **Regression / failure repair flow:** `/unit-test-failure-analyzer [targets | --log <file>]` → `/unit-test-generator --repair-failures <run>` → `/unit-test-reviewer <run>` (optional).
- **Every agent writes a JSON output file** in the target repo under `.unit-test-agent/`. The file must be human-reviewable and must be the input for the next agent.
- **Language-agnostic instructions.** Do not hard-code one ecosystem. Where ecosystem-specific detail helps (manifest files, coverage formats, mutation tools), give it as reference tables covering at least: JS/TS, Python, Java/Kotlin, C#/.NET, Go, Rust.

## The four agents

### 1. codebase-analyzer

Detect everything that can be collected automatically and write `.unit-test-agent/project.json`. It covers:

- language and runtime version, package manager, install/setup commands
- test framework and mocking library
- commands for typecheck, lint, test and coverage
- coverage report format and path
- source roots and test roots
- test file naming pattern
- 2–3 existing test files to use as style exemplars
- mutation testing tool and its command
- monorepo sub-projects
- ignore globs
- policy fields: coverage thresholds (lines, branches), `maxRepairAttempts` (default 3)

Rules:

- Each detected field records `value`, `confidence` (high / medium / low) and `evidence` (which file it came from).
- **Verify by running.** Run the detected test command on the existing suite and record the baseline result (pass/fail counts, and whether it is already failing).
- **Report gaps.** Anything that cannot be determined goes into a `missing` array. Each item is a specific question for the user and is marked `blocking: true` or `blocking: false`.
- **Never overwrite user edits.** On re-run, never overwrite a field that already has a value. Only fill nulls, refresh the evidence, and refresh `missing`.

### 2. unit-test-generator

**Generate mode** — input is targets (files, folders or globs). For each target unit:

1. Design the test cases: happy path, edge, boundary, error.
2. Write the tests following the project's framework and the exemplar style.
3. Run them and repair failures, up to `maxRepairAttempts`.
4. Run coverage scoped to the targets.

Stop condition: coverage reaches the thresholds in `project.json`. The user may set 100. Every line still uncovered must be listed with a reason (e.g. "defensive guard, unreachable from public API").

Output: `.unit-test-agent/runs/<run-id>/generation.json`. It contains:

- test files written
- test cases per file: id, target symbol, category, scenario, status
- coverage achieved vs threshold
- uncovered lines with reasons
- `suspectedDefects`
- repair attempts used

**Fix mode** (`--fix <run>`):

- Read `review.json` for that run.
- Change only what the reviewer flagged.
- Re-run the tests and coverage.
- Write `generation.r<N>.json`.

**Repair-failures mode** (`--repair-failures <run>`):

- Read `failure-analysis.json` for that run.
- Modify only the failing test files identified in the report (never modify source files):
  - For `stale_test`, `broken_mock_or_fixture`, `flaky_or_nondeterministic`, or test-level `syntax_or_import_error`: apply the recommended fix to the test (update outdated setup/mocks/assertions to match intentional API contracts, stabilize non-determinism, fix broken imports) while preserving assertion rigor.
  - For `source_regression_bug`: do **not** weaken or change the test's expected values to match buggy output. Keep the valid assertion, mark the test as skipped/expected-fail using the framework's mechanism, and record it in `suspectedDefects`.
  - For `environment_or_dependency`: do not edit test files; report the required environment/setup action as `skipped` / `blocked`.
- Re-run the repaired tests up to `maxRepairAttempts` to verify the suite passes cleanly.
- Write `.unit-test-agent/runs/<run-id>/failure-repair.json` (or `generation.repair<N>.json`) containing: repaired test files, per-failure resolution status (`fixed`, `skipped_defect`, `blocked_env`, `unresolved`), `suspectedDefects`, and repair attempts used.

### 3. unit-test-reviewer

Must run with a fresh context, separate from the generator: use a subagent where the runtime supports one. It reviews one run:

- **Review each test case against a rubric:**
  - missing branches or edge cases
  - duplicate cases
  - testing implementation details instead of behavior
  - over-mocking
  - non-determinism (time, randomness, network, filesystem)
  - weak or tautological assertions
  - naming and style versus the exemplars
- **Run mutation testing scoped to the target files only**, using the tool from `project.json` (Stryker, PIT, mutmut, cargo-mutants, etc.). If no tool is configured, report that and skip; do not fail.
- **Rerun the new tests 3 times** to detect flakiness.

Output: `.unit-test-agent/runs/<run-id>/review.json`. It contains:

- overall verdict: `approved` or `changes_requested`
- mutation score and surviving mutants (file, line, mutation)
- flaky tests
- per-case findings: case id, severity, issue, suggestion

### 4. unit-test-failure-analyzer

Analyzes failing tests from a regression suite or targeted test run and diagnoses the root cause of each failure before any repair is attempted:

- **Input modes:**
  1. **Direct execution** (`[targets]`): Run the full test command from `project.json` (or scoped to optional target files/folders/globs) and capture all failing tests and stack traces.
  2. **Log ingestion** (`--log <file>`): Parse an existing test output/log file (e.g. from CI or a prior test run) without re-running the full suite first.
- **Root-cause diagnosis:** For each failing test case, inspect the failure output, stack trace, test code, and corresponding production source code (and recent `git diff` / `git log` if available) to classify it into one of:
  - `stale_test`: Intentional production contract/signature/behavior changed, leaving the test outdated.
  - `broken_mock_or_fixture`: Test mock, stub, fixture, or setup no longer matches collaborator interfaces.
  - `source_regression_bug`: The test expectation is valid according to specs/types/contracts, and the production source code has a bug or regression.
  - `flaky_or_nondeterministic`: Failure caused by timing, randomness, shared mutable state, order dependence, or external I/O.
  - `environment_or_dependency`: Missing env var, unbuilt artifact, missing package, or runtime mismatch.
  - `syntax_or_import_error`: Compilation, type, or module resolution error in the test or source.
- **Output:** `.unit-test-agent/runs/<run-id>/failure-analysis.json`. It contains:
  - summary counts (total run, failed, analyzed, by category)
  - input source (`command` or `logFile`)
  - per-failure entries: `failureId`, `testFile`, `testName`, `targetFile`, `targetSymbol`, `errorMessage`, `stackTraceLocation`, `category`, `confidence` (`high` / `medium` / `low`), `rootCause`, `evidence` (files/lines/diff inspected), `recommendedAction` (`update_test`, `fix_mock_or_setup`, `stabilize_test`, `skip_and_report_defect`, `fix_environment`), and `proposedChangeSummary`.

## Guardrails (put these in the agent instructions)

- Write only under the configured test roots. Never modify source files.
- Never make a failing test pass by deleting or loosening an assertion, or by changing the expected value to whatever the code returned (unless failure analysis proves with high confidence that the test is `stale_test` due to an intentional contract change, and the updated assertion still strictly validates the new contract).
- If the source code looks wrong (or is classified as `source_regression_bug`), keep the correct assertion, mark the test as skipped/expected-fail using the framework's mechanism, and record it in `suspectedDefects`.
- **Check preconditions first.** Before starting, each agent confirms:
  - its input file exists
  - the input file's `schemaVersion` is supported
  - there are no blocking `missing` items

  If any check fails, stop and tell the user exactly what is needed.

- Every JSON output includes `schemaVersion`, `agent`, `createdAt` and `runId` (where applicable).

## Step 1 — Research the runtimes (do not implement yet)

Using the official documentation as of today, find out for each runtime (Claude Code, Codex CLI, Gemini CLI, GitHub Copilot in VS Code):

- where project-level custom commands, skills and agents/subagents live
- the file format and frontmatter
- how arguments are passed
- whether a fresh-context subagent is supported (needed for the reviewer)
- how users invoke it
- whether it supports the open Agent Skills format (`SKILL.md`)

Write the results to `docs/runtime-compatibility.md`: a matrix with source links and dates. Then choose a packaging approach and record it in `docs/decisions/ADR-001-packaging.md`. Aim for:

- one canonical instruction file per agent
- thin per-runtime wrappers
- no duplicated content

## Step 2 — Write the docs, then STOP for my review

1. **Feature specs** in `docs/features/`, one file per feature. Each spec covers:
   - purpose
   - invocation and arguments
   - inputs and outputs, with a complete JSON example
   - step-by-step behavior
   - edge cases
   - an acceptance-criteria checklist

   Features:
   - F01 codebase-analyzer
   - F02 generator — generate mode
   - F03 generator — fix mode
   - F04 reviewer, including mutation testing and flakiness reruns
   - F05 failure-analyzer — regression & test failure root-cause analysis (direct run or log file)
   - F06 generator — repair-failures mode (`--repair-failures <run>` based on `failure-analysis.json`)
   - F07 JSON contracts: JSON Schema files in `schemas/` for project, generation, review, failure-analysis, and failure-repair
   - F08 multi-runtime packaging and installation (copy-based, no install script)
   - F09 example fixture projects and the manual test procedure
   - F10 README / user guide
   - F11 automated output checks: a TypeScript + Vitest package in `evals/` (no UI) that checks the JSON files an agent run left behind. It only reads a run's output, so the same checks work for every runtime.
     - **Run step** (manual or nightly, never on every change): copy a fixture from `examples/` to a temp dir and run the agent chain there, either headless (`claude -p`, `codex exec`, `gemini -p`) or by hand (Copilot).
     - **Check step** (`vitest`, pointed at that temp dir), covering at least:
       - every output validates against `schemas/` and has `schemaVersion`, `agent`, `createdAt`, `runId`
       - `project.json` matches the fixture's expected values (framework, package manager, commands)
       - `git diff` shows no changes outside the test roots
       - the fixture's deliberate bug is in `suspectedDefects`, and its test is skipped/expected-fail rather than asserting the buggy value
       - the reported coverage matches what the fixture's own coverage command produces when re-run
       - every case id in `review.json` exists in `generation.json`
       - fix mode changed only flagged files and removed no assertions
       - failure-analyzer accurately classifies seeded failures (e.g. a stale test vs. a source regression bug) in `failure-analysis.json`
       - repair-failures mode fixes stale/broken tests, skips and records `source_regression_bug` items in `suspectedDefects`, and touches only failing test files
       - after a user edits a `project.json` field, re-running the analyzer keeps the edit
       - a fixture with a blocking `missing` item makes the next agent stop
     - Agent output is non-deterministic: check properties, never exact output. Report a pass rate when an agent is run more than once.

2. **Implementation plans** in `docs/plans/`, one per feature. Each plan covers:
   - files to create or change
   - ordered tasks
   - dependencies on other features
   - how to verify manually
3. **`PROGRESS.md`** at the repo root, containing:
   - a task table: feature, task, status (todo / in-progress / done / blocked), notes, links
   - per-runtime test status for each agent
   - a decisions log
   - open questions
   - a dated changelog
4. **Repo-level instructions** in `CLAUDE.md` and `AGENTS.md` describing this repo's conventions. They must tell any future session to read `PROGRESS.md` first and to resume from the next todo task.

Then stop and ask me to review. Do not implement anything until I approve.

## Step 3 — Implement (after approval)

- Work one feature at a time, in plan order.
- After each task, update `PROGRESS.md` and commit using a conventional commit message.
- Fixtures go in `examples/`: small projects in TypeScript + Vitest and in Python + pytest, with branching logic, at least one deliberate bug, and seeded failing tests for failure analysis. They are test fixtures only, not part of the agents.
- Verify each agent end-to-end against the fixtures in Claude Code. Record which runtimes have been tested and which have not in `PROGRESS.md`.
- Build F11 after F07 (schemas) and F09 (fixtures). Once it exists, record per-runtime test status from its check results.

## Proposed layout (adjust after research)

```
agents/                 canonical instructions (one per agent)
runtimes/claude/  runtimes/codex/  runtimes/gemini/  runtimes/copilot/   thin wrappers
schemas/                JSON Schema for project, generation, review, failure-analysis, failure-repair
docs/features/  docs/plans/  docs/decisions/  docs/runtime-compatibility.md
examples/               fixture projects
evals/                  automated output checks (F11, dev tooling only)
PROGRESS.md  README.md  CLAUDE.md  AGENTS.md
```

## Working rules

- If a requirement is ambiguous, ask me rather than guess. Log the question under Open questions in `PROGRESS.md`.
- Keep the instruction files concise and imperative. Agents follow short, specific rules better than long prose.
