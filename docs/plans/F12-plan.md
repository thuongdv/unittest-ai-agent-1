# Implementation Plan: F12 — Unit Test Orchestrator (`unit-test-orchestrator`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F02](F02-plan.md)–[F06](F06-plan.md): Canonical agents the orchestrator delegates to.
  - [F07: JSON Contracts](F07-plan.md): Fields the orchestrator branches on (`verdict`, `analyzedFailures`, `resolutionStatus`).

## 2. Files to Create or Change

- `agents/unit-test-orchestrator.md` — Canonical Markdown instructions for sequencing both workflows.
- Runtime wrappers are created in [F08](F08-plan.md).

## 3. Ordered Tasks

### Task F12-T1: Draft `agents/unit-test-orchestrator.md`

- **Description:** Write `agents/unit-test-orchestrator.md` covering precondition checks (stop and point to `/codebase-analyzer` when `project.json` is missing), Workflow A (generate → review → fix loop capped by `--max-review-rounds`, default 3), Workflow B (`--failures`: analyze → repair), the delegation rules table (canonical instructions, reviewer isolation, stop on step failure, branch only on JSON fields), and the final console summary.
- **Acceptance Criteria:**
  - [ ] Never invokes `codebase-analyzer` and writes no files of its own.
  - [ ] Contains zero duplicated rules from the generator, reviewer, or failure-analyzer instructions.
  - [ ] Reviewer delegation always targets the dedicated fresh-context reviewer subagent.
- **Files Touched:** `agents/unit-test-orchestrator.md`

### Task F12-T2: Verify `unit-test-orchestrator` on Fixtures

- **Description:** Run both workflows on `examples/ts-vitest` and `examples/py-pytest` and record results with `evals/` (`CHK-12`).
- **Acceptance Criteria:**
  - [ ] Workflow A produces the same files a manual chain would, within the review-round cap.
  - [ ] Workflow B classifies and repairs the seeded failures under one `<run-id>`.
  - [ ] With `project.json` removed, the orchestrator stops and no `runs/` directory is created.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. In a temp copy of `examples/ts-vitest` with `project.json` already created, run `/unit-test-orchestrator src/pricing.ts`.
2. Confirm `runs/<run-id>/` holds `generation.json`, `review.json`, and at most 2 `generation.r*.json` files, and the summary reports the final verdict.
3. Run `/unit-test-orchestrator --failures tests/regression/order.failing.ts` and confirm `failure-analysis.json` and `failure-repair.json` share one `<run-id>`.
4. Delete `.unit-test-agent/project.json`, run `/unit-test-orchestrator src/pricing.ts`, and confirm it stops with a pointer to `/codebase-analyzer`.

## 5. Risks to Verify During F08

- Generic (non-reviewer) subagent delegation differs by runtime. If a runtime cannot spawn a generic subagent, generator and analyzer steps run inline; confirm context size stays workable for 3 review rounds.
