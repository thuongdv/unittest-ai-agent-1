# F12: Unit Test Orchestrator (`unit-test-orchestrator`)

## 1. Purpose

Run a complete workflow from a single command so users do not chain agents by hand. The orchestrator only sequences the existing agents ([F02](F02-generator-generate.md)–[F06](F06-generator-repair-failures.md)), reads their JSON outputs to decide the next step, and prints a final summary. It contains no test-writing, review, or analysis logic of its own.

`codebase-analyzer` ([F01](F01-codebase-analyzer.md)) is a **one-time setup step** and is never part of an orchestrated workflow. Users re-run it by hand only when the repository's tooling changes.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation                                                                                     | Headless / CLI Invocation                                    |
| :----------------- | :--------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| **Claude Code**    | `/unit-test-orchestrator <targets> [--max-review-rounds N]`<br>`/unit-test-orchestrator --failures [targets \| --log <file>]` | `claude -p "/unit-test-orchestrator src/pricing.ts"`         |
| **Codex CLI**      | `$unit-test-orchestrator ...` (same arguments)                                                             | `codex exec "$unit-test-orchestrator src/pricing.ts"`        |
| **Gemini CLI**     | `/unit-test-orchestrator ...` (same arguments)                                                             | `gemini -p "/unit-test-orchestrator src/pricing.ts"`         |
| **GitHub Copilot** | `/unit-test-orchestrator ...` (same arguments)                                                             | N/A (VS Code Chat Agent mode)                                |

| Argument                          | Workflow           | Meaning                                                                                       |
| :-------------------------------- | :----------------- | :-------------------------------------------------------------------------------------------- |
| `<targets>`                       | Generate & review  | Files, folders, or globs passed unchanged to `unit-test-generator`.                           |
| `--max-review-rounds N`           | Generate & review  | Maximum number of reviews (default `3`). Allows up to `N - 1` fix passes.                     |
| `--failures [targets \| --log f]` | Diagnose & repair  | Arguments after `--failures` are passed unchanged to `unit-test-failure-analyzer`.            |

The individual agents remain directly invocable; the orchestrator is a convenience layer, not a replacement.

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has a supported `schemaVersion`, and has zero `blocking: true` items in `missing`.
2. If `project.json` is missing: stop immediately and tell the user to run `/codebase-analyzer` once. **Never run the analyzer automatically.**

### Outputs

- **No files of its own.** All files are written by the delegated agents, under the same `<run-id>` and with their own schemas ([F07](F07-json-contracts.md)):

| Workflow          | Files under `.unit-test-agent/runs/<run-id>/`                                   |
| :---------------- | :------------------------------------------------------------------------------ |
| Generate & review | `generation.json`, `review.json`, `generation.r1.json` … `generation.r<N-1>.json` |
| Diagnose & repair | `failure-analysis.json`, `failure-repair.json`                                  |

- **Console summary:** `run-id`, steps executed, review rounds used, final verdict or resolution counts, `suspectedDefects` count, achieved coverage, and any next action the user must take.

---

## 4. Step-by-Step Behavior

### Workflow A: Generate & Review (`<targets>`)

1. Check preconditions (Section 3).
2. Delegate `unit-test-generator <targets>`. Read the `<run-id>` it reports.
3. For `round = 1 … N`:
   1. Delegate `unit-test-reviewer <run-id>` in its dedicated **fresh-context reviewer subagent** ([F04](F04-reviewer.md)).
   2. Read `review.json`. If `verdict == "approved"`: stop, report success.
   3. If `round == N`: stop, report the outstanding findings and tell the user they can run `/unit-test-generator --fix <run-id>` by hand.
   4. Otherwise delegate `unit-test-generator --fix <run-id>` and continue.

### Workflow B: Diagnose & Repair (`--failures ...`)

1. Check preconditions (Section 3).
2. Delegate `unit-test-failure-analyzer` with the arguments after `--failures`. Read the `<run-id>` it reports.
3. Read `failure-analysis.json`. If `analyzedFailures == 0`: stop, report that all tests pass.
4. Delegate `unit-test-generator --repair-failures <run-id>`.
5. Read `failure-repair.json` and report counts by `resolutionStatus` (`fixed`, `skipped_defect`, `blocked_env`, `unresolved`), listing every `suspectedDefects` entry and every `blocked_env` step the user must take.

### Delegation Rules

| Rule                     | Requirement                                                                                                                  |
| :----------------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| Canonical instructions   | Each step follows `.unit-test-agent/agents/<agent>.md` with the forwarded arguments. Never restate or override agent rules.   |
| Reviewer isolation       | The reviewer always runs in its dedicated subagent, never in a context that ran the generator.                                |
| Other steps              | Run in a subagent where the runtime supports it; otherwise run inline.                                                       |
| Step failure             | If a step stops (failed precondition, missing expected output file, unsupported `schemaVersion`), stop the workflow and report which step failed and why. Never skip ahead. |
| Decisions                | Base every branch only on fields in the JSON outputs (`verdict`, `analyzedFailures`, `resolutionStatus`), never on console text. |

---

## 5. Edge Cases

- **`project.json` missing:** Stop before creating any `runs/` directory; instruct the user to run `/codebase-analyzer`.
- **Blocking `missing` item:** Stop and list the blocking questions; do not delegate any step.
- **Reviewer approves on the first round:** No `generation.r*.json` is written.
- **`--max-review-rounds 1`:** Generate once, review once, never fix.
- **Generator leaves tests with `status: "failed"`:** Continue to review; the reviewer reports them.
- **`--failures` with every failure `blocked_env`:** Report the environment steps; no test files change.

---

## 6. Acceptance Criteria Checklist

- [ ] Stops with a clear message and writes nothing when `project.json` is missing, unsupported, or has a `blocking: true` item.
- [ ] Never invokes `codebase-analyzer`.
- [ ] Workflow A stops on the first `approved` verdict and never exceeds `N` reviews or `N - 1` fix passes.
- [ ] Workflow B runs analyze → repair under one `<run-id>` and skips repair when there are zero failures.
- [ ] The reviewer always runs in its fresh-context subagent.
- [ ] Writes no files itself; all outputs come from delegated agents and validate against their schemas.
- [ ] Contains no test-writing, review, or classification rules (all such rules stay in `agents/unit-test-generator.md`, `unit-test-reviewer.md`, `unit-test-failure-analyzer.md`).
